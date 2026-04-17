# 第 24 关：错误处理（师兄带你学 Go）

## 🎯 这一关你会学到

本关带你搞懂 Go 的错误处理哲学：错误不是异常，而是值。Go 没有 try-catch，而是借助 `error` 接口与多返回值，把错误像普通数据一样在函数间传递。掌握这套机制，你就能写出清晰、可控、符合 Go 风格的错误处理代码。

## 🤔 先想一个问题

想象一位外卖骑手到了你门口，发现汤洒了。"XX 小哥那单汤洒了，需要重新配餐"，让调度中心决定怎么处理——重配、退款、还是换个餐厅。整个过程错误是一种"值"，被显式传递出去，而不是炸锅式地抛出。Go 的错误处理跟这个一模一样：**把错误当成普通返回值**往调用栈上汇报，由调用方决定如何处理。

**在 Go 里错误是一种值——任何实现了 `error` 接口（`Error() string`）的类型都是错误。函数把错误作为最后一个返回值返回，调用方用 `if err != nil` 显式检查。**

## 📖 看代码

```go
// 符合 Go 语言习惯的做法是使用一个独立、明确的返回值来传递错误信息。
// 这与 Java、Ruby 使用的异常（exception）
// 以及在 C 语言中有时用到的重载 (overloaded) 的单返回/错误值有着明显的不同。
// Go 语言的处理方式能清楚的知道哪个函数返回了错误，并使用跟其他（无异常处理的）语言类似的方式来处理错误。

package main

import (
	"errors"
	"fmt"
)

// 按照惯例，错误通常是最后一个返回值并且是 `error` 类型，它是一个内建的接口。
func f1(arg int) (int, error) {
	if arg == 42 {

		// `errors.New` 使用给定的错误信息构造一个基本的 `error` 值。
		return -1, errors.New("can't work with 42")

	}

	// 返回错误值为 nil 代表没有错误。
	return arg + 3, nil
}

// 你还可以通过实现 `Error()` 方法来自定义 `error` 类型。
// 这里使用自定义错误类型来表示上面例子中的参数错误。
type argError struct {
	arg  int
	prob string
}

func (e *argError) Error() string {
	return fmt.Sprintf("%d - %s", e.arg, e.prob)
}

func f2(arg int) (int, error) {
	if arg == 42 {

		// 在这个例子中，我们使用 `&argError` 语法来建立一个新的结构体，
		// 并提供了 `arg` 和 `prob` 两个字段的值。
		return -1, &argError{arg, "can't work with it"}
	}
	return arg + 3, nil
}

func main() {

	// 下面的两个循环测试了每一个会返回错误的函数。
	// 注意，在 `if` 的同一行进行错误检查，是 Go 代码中的一种常见用法。
	for _, i := range []int{7, 42} {
		if r, e := f1(i); e != nil {
			fmt.Println("f1 failed:", e)
		} else {
			fmt.Println("f1 worked:", r)
		}
	}
	for _, i := range []int{7, 42} {
		if r, e := f2(i); e != nil {
			fmt.Println("f2 failed:", e)
		} else {
			fmt.Println("f2 worked:", r)
		}
	}

	// 如果你想在程序中使用自定义错误类型的数据，
	// 你需要通过类型断言来得到这个自定义错误类型的实例。
	_, e := f2(42)
	if ae, ok := e.(*argError); ok {
		fmt.Println(ae.arg)
		fmt.Println(ae.prob)
	}
}
```

## 🔍 师兄给你逐行拆

### error 接口：错误是值

`error` 是 Go 的内置接口，定义极简，只有一个方法：

```go
type error interface {
    Error() string
}
```

任何实现了 `Error() string` 的类型都满足 `error` 接口。`errors.New("...")` 可以快速创建一个最基础的错误值：

```go
err := errors.New("文件不存在")
```

Go 的惯例是：函数返回 `(结果, error)`，错误放在最后一个返回值。调用方用 `if err != nil` 显式检查：

```go
result, err := doSomething()
if err != nil {
    // 处理错误
}
```

这比 try-catch 的优势在于：**错误路径显式、控制流直观**，你一眼就能看到哪里可能出错、如何处理。编译器也会帮你把关——若用 `_` 丢弃了 `err`，静态检查工具（如 `errcheck`）会发出警告，防止错误被静默忽略。

> 记住：在 Go 里，错误不是异常，而是普通的值，和其他返回值平起平坐。

### 自定义错误类型：带上更多上下文

Go 的 `error` 只是一个接口，只要实现 `Error() string` 方法，任何结构体都能成为错误类型。

```go
type argError struct {
    arg  int
    prob string
}

func (e *argError) Error() string {
    return fmt.Sprintf("%d - %s", e.arg, e.prob)
}
```

返回时用 `&argError{arg, prob}` 即可。

**真正的好处在调用方**。用 `errors.New` 返回的错误，调用方只能拿到一段字符串，无法进一步区分细节。而自定义类型可以通过**类型断言**还原出原始结构：

```go
if aerr, ok := err.(*argError); ok {
    fmt.Println(aerr.arg)  // 拿到具体字段，做精细处理
}
```

这样，调用方可以根据 `aerr.arg` 的值走不同的分支逻辑，而不是靠字符串匹配——既健壮又不怕消息格式改变。需要携带多个字段时，直接往结构体里加即可，扩展成本极低。

### if err := f(); err != nil 的常见姿势

Go 里最高频的错误处理模式长这样：

```go
if r, e := f1(i); e != nil {
    fmt.Println("failed:", e)
} else {
    fmt.Println("worked:", r)
}
```

核心点：变量声明和错误检查写在同一行 `if` 语句的初始化部分，`r` 和 `e` 的**作用域仅限这个 if/else 块**，出了大括号就失效，不会污染外层命名空间，干净整洁。

这个模式在 Go 代码里几乎无处不在——打开文件、发 HTTP 请求、解析 JSON，每一步都来一遍。Go 社区有个经久不衰的梗：新手打开一份 Go 源码，扑面而来全是 `if err != nil`，密密麻麻，视觉上略显啰嗦。

但换个角度看，这恰恰是 Go 的哲学所在：**错误路径显式可见**。调用方必须亲手处理或转发每一个错误，不可能被悄悄吞掉。相比 Java 的 checked exception——异常在调用栈里层层传递，中间究竟经过哪些路径往往难以追踪——Go 的方式反而更透明、更可控。啰嗦是代价，但清晰是收益。

### 进阶：errors.Is / errors.As / %w 包装

Go 1.13 引入了错误包装机制，彻底改变了多层错误处理的方式。

**`%w` 包装错误**

```go
fmt.Errorf("read config: %w", err)
```

这会把原始 `err` 嵌入新错误中，形成一条**错误链**，同时保留完整的上下文信息。

**`errors.Is` — 判断哨兵错误**

```go
if errors.Is(err, io.EOF) {
    // 错误链中任意一层等于 io.EOF
}
```

不再需要 `err == io.EOF`，即使错误被层层包装，`Is` 依然能穿透链条找到目标。

**`errors.As` — 提取具体类型**

```go
var pathErr *fs.PathError
if errors.As(err, &pathErr) {
    fmt.Println(pathErr.Path)
}
```

`As` 沿错误链向下查找，找到第一个可赋值给目标类型的层，并将其提取出来，方便访问结构体字段。

**设计哲学**

三者配合形成分层惯例：底层函数用 `%w` 包装细节，中层添加业务上下文，顶层用 `Is` / `As` 按类型或值做决策。比裸字符串比较健壮得多，也是现代 Go 错误处理的标配写法。

## 🏃 跑一下试试

```bash
$ go run errors.go
f1 worked: 10
f1 failed: can't work with 42
f2 worked: 10
f2 failed: 42 - can't work with it
42
can't work with it
```

最后两行是 `main` 里对 `*argError` 做了类型断言后，单独取出 `arg` 和 `prob` 字段的演示输出。

## 💡 师兄的碎碎念

- 用 `go vet` 配合 `errcheck` 静态检查被忽略的 `err` 返回值，避免静默吞错
- 优先用 `errors.Is` 而非 `err == sentinel`，前者能穿透 `%w` 包装的错误链
- `fmt.Errorf("...: %w", err)` 包装并保留原始错误链；若只需打印信息用 `%v`，不保留链
- 返回 `error` 时同时返回的结果值应视为无效，调用方务必先检查 `err != nil` 再使用结果

## 🎓 这一关的知识点清单

- **error 接口**：Go 内置接口，唯一方法是 `Error() string`，任何实现它的类型都可作为错误。
- **errors.New / fmt.Errorf**：快速构造带消息的错误值；`fmt.Errorf("ctx: %w", err)` 还能把底层错误包装进链。
- **自定义 error**：把 error 实现在自定义 struct 上，携带 `arg`/`code` 等结构化信息，供调用方按类型断言分支处理。
- **`if err != nil` 模式**：Go 最常见的错误检查写法，错误路径显式可见，不会被静默吞掉。
- **errors.Is / errors.As**：Go 1.13+ 标准姿势，`Is` 判断是否匹配哨兵错误，`As` 提取错误链中的具体类型，替代脆弱的字符串比较。

## ➡️ 下一关

继续前进，下一关进入并发世界！

[下一关：Goroutines →](../25-goroutines/)
