# 第 19 关：结构体（师兄带你学 Go）

## 🎯 这一关你会学到

本关目标：搞懂 struct 的完整用法。首先学会用 `type` 关键字定义结构体；然后掌握三种实例化方式——按位置、按字段名、省略零值字段；接着学习如何封装构造函数统一创建实例；最后理解通过 `.` 访问字段，以及值类型与指针类型在传参和修改时的本质区别。

## 🤔 先想一个问题

想象宿舍门口贴着一张信息卡，上面写着同学的姓名、年龄和学号。这三条信息描述的是同一个人，但如果你用三个独立变量分别存储——`name`、`age`、`studentID`——一旦同宿舍住了好几个人，变量就会爆炸式增长，彼此之间的关联也会变得混乱不堪。

**结构体 struct 是把若干命名字段打包成整体的数据类型；用 `type 名字 struct { 字段 类型 }` 定义，用 `名字{字段值}` 创建实例。**

## 📖 看代码

```go
// Go 的_结构体(struct)_ 是带类型的字段(fields)集合。
// 这在组织数据时非常有用。

package main

import "fmt"

// 这里的 `person` 结构体包含了 `name` 和 `age` 两个字段。
type person struct {
	name string
	age  int
}

// `newPerson` 使用给定的名字构造一个新的 person 结构体.
func newPerson(name string) *person {
	// 您可以安全地返回指向局部变量的指针，
	// 因为局部变量将在函数的作用域中继续存在。
	p := person{name: name}
	p.age = 42
	return &p
}

func main() {

	// 使用这个语法创建新的结构体元素。
	fmt.Println(person{"Bob", 20})

	// 你可以在初始化一个结构体元素时指定字段名字。
	fmt.Println(person{name: "Alice", age: 30})

	// 省略的字段将被初始化为零值。
	fmt.Println(person{name: "Fred"})

	// `&` 前缀生成一个结构体指针。
	fmt.Println(&person{name: "Ann", age: 40})

	// 在构造函数中封装创建新的结构实例是一种习惯用法
	fmt.Println(newPerson("Jon"))

	// 使用`.`来访问结构体字段。
	s := person{name: "Sean", age: 50}
	fmt.Println(s.name)

	// 也可以对结构体指针使用`.` - 指针会被自动解引用。
	sp := &s
	fmt.Println(sp.age)

	// 结构体是可变(mutable)的。
	sp.age = 51
	fmt.Println(sp.age)
}
```

## 🔍 师兄给你逐行拆

### 定义 struct 与按位置初始化

使用 `type` 关键字可以定义一个自定义结构体类型：

```go
type person struct {
    name string
    age  int
}
```

这里定义了一个名为 `person` 的类型，包含两个字段：`name` 和 `age`。

**按位置初始化**是最简洁的写法，按照字段的定义顺序依次传值：

```go
p := person{"Bob", 20}
fmt.Println(p) // 输出：{Bob 20}
```

Go 会将 `"Bob"` 赋给第一个字段 `name`，将 `20` 赋给第二个字段 `age`。

⚠️ **坑：依赖字段顺序**

位置初始化与字段的定义顺序强绑定。如果日后有人在 struct 中调整了字段顺序，或者插入了新字段，所有使用位置初始化的代码都会悄悄出错——编译器不一定报错，但值会跑错字段，产生难以排查的 bug。

因此，在实际项目中更推荐使用**具名初始化**（`person{name: "Bob", age: 20}`），可读性更好，也更安全。位置初始化通常只在字段极少且结构稳定的内部代码中使用。

### 按字段名初始化 + 省略字段用零值

在初始化结构体时，推荐使用 **字段名: 值** 的形式：

```go
p1 := person{name: "Alice", age: 30}
p2 := person{name: "Fred"}  // age 省略，自动为零值 0
```

`p2.age` 的值是 `0`，因为 Go 保证每个未显式赋值的字段都会被初始化为该类型的**零值**：

- `string` → `""`
- `int` → `0`
- `bool` → `false`

**为什么优先用字段名初始化？**

1. **更安全**：即使以后结构体新增或调整了字段顺序，代码也不会出错。
2. **更清晰**：读代码时一眼就知道每个值对应哪个字段，无需对照结构体定义。
3. **省略即零值**：不需要某个字段时直接省略，Go 自动补上安全的默认值，无需手动写 `0` 或 `""`。

总结：字段名初始化是 Go 社区的惯用做法，写库或团队项目时尤为重要。

### 取指针 & 和构造函数 newPerson

在 `struct` 字面量前加 `&`，可以直接拿到一个指向该结构体的指针（`*person`），省去先声明再取地址的步骤：

```go
p := &person{name: "Alice", age: 30} // 类型是 *person
```

更常见的做法是封装一个**构造函数**：

```go
func newPerson(name string) *person {
    p := person{name: name}
    p.age = 18
    return &p // 返回局部变量的地址
}
```

初学者可能担心：函数返回后 `p` 不是销毁了吗？答案是不会。Go 编译器有**逃逸分析**机制，一旦发现局部变量的地址会逃逸到函数外部，就自动将其分配到堆上，调用方拿到的指针始终有效，完全安全。

**生活类比**：构造函数就像工厂的流水线——每次调用都按同一套规范生产出一个合格的实例，外部无需关心内部初始化细节，只管拿来即用。

### 访问字段：. 同时兼容值和指针

通过 `.` 运算符读写字段：

```go
fmt.Println(s.name) // 读
s.age = 51          // 写
```

Go 最甜的语法糖之一：**值和指针使用完全相同的 `.` 语法**，编译器自动完成解引用。

```go
sp := &s
sp.age = 51      // 等价于
(*sp).age = 51   // 无需像 C 那样写 sp->age
```

无论变量是 `Person` 还是 `*Person`，一律用 `.`，代码更简洁，也不容易出错。

**struct 的比较**：当 struct 的所有字段类型都支持 `==` 时，两个 struct 可以直接用 `==` 比较——逐字段对比，全部相等才为 `true`。

```go
a := Point{1, 2}
b := Point{1, 2}
fmt.Println(a == b) // true
```

若任意字段是 slice、map 等不可比较类型，则编译直接报错，而非留到运行时。

## 🏃 跑一下试试

```bash
$ go run structs.go
{Bob 20}
{Alice 30}
{Fred 0}
&{Ann 40}
&{Jon 42}
Sean
50
51
```
其中 `{…}` 表示按值打印 struct，`&{…}` 表示打印的是指针（前缀 `&` 标识地址语义）。

## 💡 师兄的碎碎念

- 优先使用**字段名初始化** `Person{Name: "Bob", Age: 20}`，避免字段顺序变动引发隐蔽 bug。
- `&T{}` 是创建 struct 指针的惯用写法，等价于 `new(T)` 再赋值，日常代码中更常见。
- 只有**首字母大写**的字段（exported）才能在其他包中访问；包内私有字段用小写开头。
- 将 struct 用作 **map key** 时，要求其所有字段类型均为 comparable，含 slice/map 的 struct 不可作为 key。

## 🎓 这一关的知识点清单

- **struct 定义**：用 `type Name struct { Field Type }` 声明自定义复合类型，字段按顺序排列。
- **字段初始化方式**：可按位置 `{val1, val2}` 或按名称 `{Field: val}` 初始化，推荐后者更清晰。
- **零值机制**：未赋值的字段自动获得该类型零值（int→0，string→""，bool→false）。
- **指针自动解引用**：通过 struct 指针访问字段时，Go 自动解引用，`p.Field` 等价于 `(*p).Field`。
- **struct 相等性**：两个 struct 可用 `==` 比较，前提是所有字段类型均为 comparable（不含 slice/map/func）。

## ➡️ 下一关

掌握 struct 之后，下一步学习如何为类型绑定行为方法。[下一关：Methods →](../20-methods/)
