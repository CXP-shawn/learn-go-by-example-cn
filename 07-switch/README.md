# 第 7 关：Switch 分支结构（师兄带你学 Go）

## 🎯 这一关你会学到

这一关咱们学 Go 里的 switch 分支结构，包括基本用法、多值 case、不带表达式的 switch，以及最酷的「类型开关」。学完之后，你能用 switch 优雅替代一堆 if-else，写出更清晰的多分支逻辑，面试也常考！

## 🤔 先想一个问题

想象你去奶茶店点单，店员问你要几分糖：「你说1分，她给你无糖；你说3分，她给你少糖；你说5分，给标准糖；啥都没说，默认全糖。」这其实就是程序里的 **switch 语句**——根据一个值，匹配不同的分支，执行对应的操作。

准确的技术定义：**switch 语句**是一种多路分支控制结构，它对一个表达式求值，然后将结果与多个 **case 子句**逐一比较，执行第一个匹配成功的 case 块；若所有 case 都不匹配，则执行可选的 **default 子句**。相比写一堆 if-else if-else，switch 让代码可读性更强、结构更清晰，Go 的 switch 还自带「不穿透」特性，比 C/Java 更安全！

## 📖 看代码

```go
// _switch_ 是多分支情况时快捷的条件语句。

package main

import (
 "fmt"
 "time"
)

func main() {

 // 一个基本的 `switch`。
 i := 2
 fmt.Print("write ", i, " as ")
 switch i {
 case 1:
 fmt.Println("one")
 case 2:
 fmt.Println("two")
 case 3:
 fmt.Println("three")
 }

 // 在同一个 `case` 语句中，你可以使用逗号来分隔多个表达式。
 // 在这个例子中，我们还使用了可选的 `default` 分支。
 switch time.Now().Weekday() {
 case time.Saturday, time.Sunday:
 fmt.Println("It's the weekend")
 default:
 fmt.Println("It's a weekday")
 }

 // 不带表达式的 `switch` 是实现 if/else 逻辑的另一种方式。
 // 这里还展示了 `case` 表达式也可以不使用常量。
 t := time.Now()
 switch {
 case t.Hour() < 12:
 fmt.Println("It's before noon")
 default:
 fmt.Println("It's after noon")
 }

 // 类型开关 (`type switch`) 比较类型而非值。可以用来发现一个接口值的类型。
 // 在这个例子中，变量 `t` 在每个分支中会有相应的类型。
 whatAmI := func(i interface{}) {
 switch t := i.(type) {
 case bool:
 fmt.Println("I'm a bool")
 case int:
 fmt.Println("I'm an int")
 default:
 fmt.Printf("Don't know type %T
", t)
 }
 }
 whatAmI(true)
 whatAmI(1)
 whatAmI("hey")
}
```

## 🔍 师兄给你逐行拆

### 第一段：基本 switch——按值匹配

```go
i := 2
fmt.Print("write ", i, " as ")
switch i {
case 1:
    fmt.Println("one")
case 2:
    fmt.Println("two")
case 3:
    fmt.Println("three")
}
```

咱们先从最简单的 switch 开始。这里 `i := 2`，声明了一个整数变量，然后 `switch i` 的意思是：「拿 i 的值去挨个 case 比对，谁匹配上就执行谁。」

这里 i 等于 2，所以直接命中 `case 2:`，输出 `two`，其他 case 压根不看。

**师兄要重点说一个 Go 的坑——Go 的 switch 默认不穿透（no fallthrough）！** 在 C 或 Java 里，switch 的每个 case 如果不写 `break`，会一路向下执行所有 case，这是经典 bug 来源。Go 直接把这个行为反过来了：每个 case 执行完自动 break，除非你手动写 `fallthrough` 关键字才会继续往下走。所以 Go 的 switch 天生更安全，不容易写出「忘加 break」的低级错误。

生活类比：就像你去便利店扫码支付，扫完就结束了，不会再扫第二次。不像老式收银机需要你手动按「确认」结束，Go 帮你省了这一步。

注意 `fmt.Print` 和 `fmt.Println` 的区别：`Print` 不换行，`Println` 自动换行。所以输出是 `write 2 as two`，在同一行拼起来的。

---

### 第二段：多值 case + default——一个 case 匹配多个值

```go
switch time.Now().Weekday() {
case time.Saturday, time.Sunday:
    fmt.Println("It's the weekend")
default:
    fmt.Println("It's a weekday")
}
```

这段代码获取今天是星期几，然后判断是否是周末。

`time.Now()` 返回当前时间，`.Weekday()` 方法返回一个 **`time.Weekday` 类型**的值，它本质上是一个整数枚举，`time.Saturday` 和 `time.Sunday` 是预定义的常量。

重点来了：`case time.Saturday, time.Sunday:` ——一个 case 里用**逗号分隔了两个值**！意思是「只要是周六或者周日，就执行这个 case」。这个写法在很多语言里需要用 `||` 或者写两个 case，Go 直接支持逗号多值，相当优雅。

`default:` 是**默认分支**，当所有 case 都没有匹配时执行。这里的逻辑是：不是周末，那就是工作日，输出 `It's a weekday`。

生活类比：就像宿舍门禁系统，你刷脸进门，系统匹配「张三、李四」这两个人允许在规定时间外进入，其他人一律按普通规则处理（default）。一个规则覆盖多个人，不用给每个人单独写一条。

踩坑提示：`default` 不一定要放在最后面，Go 允许你把 `default` 写在任意位置，但为了可读性，师兄建议还是放末尾，别给队友挖坑。

---

### 第三段：不带表达式的 switch——当 if-else 的高级替代品

```go
t := time.Now()
switch {
case t.Hour() < 12:
    fmt.Println("It's before noon")
default:
    fmt.Println("It's after noon")
}
```

这里 `switch` 后面没有跟任何变量或表达式，直接跟 `{`，这就是 Go 的**无表达式 switch**（也叫 expressionless switch）。

没有表达式的 switch，每个 case 后面可以写**任意布尔表达式**，Go 会从上到下逐一求值，第一个结果为 `true` 的 case 被执行。这本质上就是 if-else if-else 的另一种写法，但视觉上更整齐。

`t.Hour()` 返回当前小时数（0-23），如果小于 12 就是上午，否则是下午或晚上。

为什么要用无表达式 switch 而不是 if-else？说实话，功能上完全等价，只是风格不同。当你有三个以上的条件分支时，switch 写法会比一堆 `else if` 更清晰。Go 官方代码库里也经常见到这种写法，算是 Go 圈子的惯用法（idiom）。

注意：这里变量名 `t` 被重新赋值为 `time.Now()`，前面的 `i` 已经没用了，Go 不允许声明了变量不用（编译报错），所以每个变量都要有意义地使用。

生活类比：就像你早上出门前看天气，不用说「如果今天是星期几」，直接判断「气温低于10度就穿羽绒服，低于20度穿外套，否则短袖」，每个条件都是独立的布尔判断。

---

### 第四段：类型开关（type switch）——Go 最特别的技能之一

```go
whatAmI := func(i interface{}) {
    switch t := i.(type) {
    case bool:
        fmt.Println("I'm a bool")
    case int:
        fmt.Println("I'm an int")
    default:
        fmt.Printf("Don't know type %T\n", t)
    }
}
whatAmI(true)
whatAmI(1)
whatAmI("hey")
```

这段是本关最烧脑也最实用的部分——**类型开关（type switch）**。

先解释几个概念：

- **`interface{}`**：Go 里的空接口类型，可以接收任意类型的值。相当于「万能容器」，不管你传 int、string、bool、struct，它都能装。在 Go 1.18 以后也可以写成 `any`，是个别名。
- **类型断言 `i.(type)`**：这个语法只能用在 switch 里，作用是「取出 i 里面实际存储的值的类型」，然后和各个 case 的类型进行比较。这里的 `t` 是一个新变量，在每个 case 分支里，`t` 的类型就是该 case 对应的具体类型，编译器会自动帮你做类型转换。

`whatAmI` 是一个**匿名函数**，赋值给变量 `whatAmI`，然后通过 `whatAmI(true)` 这样的方式调用。这也是 Go 里函数是「一等公民」的体现。

执行流程：
- `whatAmI(true)`：传入 bool 值，命中 `case bool:`，输出 `I'm a bool`
- `whatAmI(1)`：传入 int 值，命中 `case int:`，输出 `I'm an int`
- `whatAmI("hey")`：传入 string 值，没有对应 case，走 `default:`，`%T` 是格式化动词，输出变量的实际类型名，所以输出 `Don't know type string`

生活类比：就像快递柜收到一个包裹，系统先扫描条码，判断是「冷链包裹」还是「普通快递」还是「大件货」，然后分别放到不同的格子里处理。类型开关就是在运行时动态识别「包裹的种类」。

踩坑区：`i.(type)` 这个语法**只能在 switch 里用**！你要是在普通代码里写 `x := i.(type)`，编译器直接报错。如果想在 switch 外面做类型断言，要用 `x, ok := i.(int)` 这种形式，ok 表示断言是否成功。这是两种不同的语法，别搞混。

另外，`fmt.Printf` 里的 `%T` 是打印变量**类型名**的格式化占位符，不是打印值，这个在调试时候特别好用，师兄经常用它来看一个变量到底是什么类型。

---

### 总结一下这关的核心知识点

1. **基本 switch**：`switch 变量 { case 值: ... }` 按值匹配，自动 break 不穿透
2. **多值 case**：一个 case 里用逗号写多个值，覆盖多种情况
3. **default 分支**：兜底处理，所有 case 都不匹配时执行
4. **无表达式 switch**：`switch { case 布尔表达式: ... }` 是 if-else 的优雅替代
5. **类型开关**：`switch t := i.(type) { case 某类型: ... }` 在运行时判断接口值的实际类型

Go 的 switch 比 C/Java 更安全（默认不穿透），比 Python 的 match（3.10才加）早了很多年，是日常开发里非常高频的结构，值得把这几种写法都烂熟于心！

## 🏃 跑一下试试

```bash
go run switch.go
Write 2 as two
It's a weekday
It's after noon
I'm a bool
I'm an int
Don't know type string
```

## 💡 师兄的碎碎念

- Go 的 `switch` 不需要写 `break`，这和 C/Java 不一样，刚转过来的同学容易手滑多写一个 😅
- 想让某个 `case` 故意穿透到下一个？用 `fallthrough` 关键字，但现实中用到的场景真的很少，别滥用
- 类型开关里那个 `i.(type)` 语法只能在 `switch` 里用，单独写 `i.(type)` 会直接报编译错误，别踩坑
- `time.Now().Weekday()` 这种用法很常见，`switch` 搭配返回值直接用，比提前赋变量更简洁

## 🎓 这一关的知识点清单

- **基础 switch**：`switch` 根据变量值匹配 `case`，无需写 `break`，Go 默认不穿透
- **多值 case**：同一个 `case` 可以用逗号隔开多个值，比写多个 `case` 优雅多了
- **无表达式 switch**：`switch {}` 省略条件，相当于更清爽的 `if / else if / else` 链
- **类型开关 type switch**：`switch t := i.(type)` 可以判断接口变量的底层类型，处理多态场景必备
- **不穿透原则**：Go 的 `switch` 默认每个 `case` 执行完就退出，想穿透要显式用 `fallthrough`

## ➡️ 下一关

数组是 Go 里存一组同类型数据的基础结构，下一关我们来看看它怎么用 👉 [下一关：数组 →](../08-arrays/)
