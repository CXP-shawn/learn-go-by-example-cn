# 第 46 关：Defer（师兄带你学 Go）

🎯 这一关你会学到
• 什么是 defer——把一个函数调用"延迟"到当前函数返回前执行
• defer 的典型用途：清理资源（关文件、解锁、关连接）
• defer 执行顺序：后进先出（LIFO），像往弹夹里塞子弹
• defer 参数的"立即求值"机制，新手最容易踩的坑

🤔 先想一个问题

你下班点个外卖，到家那一刻你会干啥？放下袋子、拆袋、吃饭、收拾垃圾——**收拾垃圾**这步其实在你下单那一刻就"预订"好了，只是执行时机被推到最后。Go 的 defer 就是干这事儿的：你打开文件的那一刻，就顺手把"最后记得关"这件事告诉编译器，编译器会在函数返回前自动替你执行。这样你就不用担心中途写一半想起来要关文件、或者因为半路 panic 了没人收尾。这一关咱们就看看 defer 怎么把"资源清理"这件事变成肌肉记忆。

## 📖 看代码

```go
// _Defer_ 用于确保程序在执行完成后，会调用某个函数，一般是执行清理工作。
// Defer 的用途跟其他语言的 `ensure` 或 `finally` 类似。

package main

import (
	"fmt"
	"os"
)

// 假设我们想要创建一个文件、然后写入数据、最后在程序结束时关闭该文件。
// 这里展示了如何通过 `defer` 来做到这一切。
func main() {

	// 在 `createFile` 后立即得到一个文件对象，
	// 我们使用 defer 通过 `closeFile` 来关闭这个文件。
	// 这会在封闭函数（`main`）结束时执行，即 `writeFile` 完成以后。
	f := createFile("/tmp/defer.txt")
	defer closeFile(f)
	writeFile(f)
}

func createFile(p string) *os.File {
	fmt.Println("creating")
	f, err := os.Create(p)
	if err != nil {
		panic(err)
	}
	return f
}

func writeFile(f *os.File) {
	fmt.Println("writing")
	fmt.Fprintln(f, "data")

}

func closeFile(f *os.File) {
	fmt.Println("closing")
	err := f.Close()
	// 关闭文件时，进行错误检查是非常重要的，
	// 即使在 defer 函数中也是如此。
	if err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}
}
```

🔍 师兄给你逐行拆

**defer 是啥？先把心智模型建好**

defer 是 Go 的关键字（keyword，编译器直接识别的保留词，和 if、for 一个级别）。它做的事情就一句话：**把后面跟着的函数调用登记起来，等当前函数快要返回的时候再执行**。注意是"登记"，不是"立刻执行"——defer 这一行写完就走了，被 defer 的那个函数要等到外层函数 return 前那一刻才会真的跑起来。

举个生活里的类比：你去便利店买东西，店员扫码后说"小票需要吗？需要的话我打印"，你说"要，先放收银台吧，我出门前拿"——defer 就是这股劲儿，登记完任务先忙别的，最后结账前再把登记过的活儿挨个做完。

**main 函数里 defer 的位置玄机**

重点看 main 这几行：

```go
f := createFile("/tmp/defer.txt")
defer closeFile(f)
writeFile(f)
```

咱们逐行拆：

第一行，createFile 创建了一个叫 /tmp/defer.txt 的文件，返回一个 *os.File 指针（指针（pointer）的定义是"一个变量，它的值是另一个变量在内存里的地址"，`*os.File` 表示"指向 os.File 对象的指针"）。createFile 里先打印 "creating"，然后调 os.Create 真正创建文件；如果出错直接 panic，这里假定不会出错。

第二行，`defer closeFile(f)` ——这是整关的灵魂。这一行**并不会现在就调用 closeFile**，而是告诉 Go 运行时："给我记住，等 main 快 return 的时候，帮我执行 closeFile(f)。" 注意 `f` 这个参数是**当场求值**的，不是等到真执行时再算——所以现在就决定了 defer 时带着的是这个具体的文件指针。

第三行 writeFile(f) 是立刻执行的，往文件里写了一行 "data"。

然后 main 函数走完 writeFile 返回，正要 return 之前，Go 运行时把之前登记的 closeFile(f) 挨个跑一遍。这时候才真的打印 "closing"、关闭文件。

**defer 的执行顺序：后进先出（LIFO）**

师兄在这里提一个重要规则：同一个函数里可以写好多 defer，它们的执行顺序是**后登记的先执行**，也就是 **LIFO（Last In First Out，后进先出）**。你可以把它想象成往弹夹里塞子弹：最后塞进去的子弹在最上面，开枪时最先打出去。或者更接地气的——你往书包里塞书，最后塞进去的放在最上面，掏出来肯定是它先被掏到。本关代码只有一个 defer 看不出来，但一旦你 defer 三个清理动作，顺序一定是反着来。

**createFile / writeFile / closeFile 三兄弟都干啥**

- `createFile`：打印 "creating"，用 os.Create 创建文件，出错就 panic，正常返回 *os.File 给调用方
- `writeFile`：打印 "writing"，用 fmt.Fprintln 往文件里写了一行 "data"（Fprintln 定义：把内容写到第一个参数给的 io.Writer 里，末尾自动加换行）
- `closeFile`：打印 "closing"，调 f.Close() 真正关闭文件。**特别注意**它里面还做了错误检查——因为 Close 也可能出错（磁盘满、网络文件系统超时都会让 Close 报错），真出错了用 fmt.Fprintf 写到 os.Stderr 然后 os.Exit(1) 非正常退出。

**为啥 Close 这么一个"小"操作还要检查错误？**

很多新人觉得 Close 就是收尾，出错也无所谓。错！你写的 "data" 可能还在操作系统的缓冲区里没真正落盘，Close 时才触发真正的写磁盘——这时磁盘满了、掉电了，Close 就会给你返回 error。如果不检查，你以为写成功了实际上数据丢了，这种 bug 贼难排查。所以 Go 社区的建议是：**即使在 defer 里，也要检查 Close 返回的错误**。

**defer 和 panic 的协作**

还记得上一关的 panic 吗？panic 触发栈展开（unwinding）时，会沿途执行每一层已经登记的 defer。所以把关文件、解锁这种"必须做"的动作放 defer 里是**panic-safe** 的——哪怕中间炸了，defer 照样执行。这就是为啥 Go 社区里 defer + panic 组合是一种广泛使用的"资源安全"模式。

🏃 跑一下试试

保存为 defer.go，然后：

```bash
go run defer.go
```

预期输出：

```text
creating
writing
closing
```

逐行看：
- `creating` —— main 里先调 createFile，它先打这行
- `writing` —— 然后 writeFile 真正往文件里写 "data"
- `closing` —— 最后 main 快 return 了，被 defer 的 closeFile 才跑起来，打这行

虽然代码里 defer closeFile 写在 writeFile 之前，但它的执行时机被推到了 writeFile 之后、main 返回之前——这就是 defer 的魔法。

💡 师兄的碎碎念

defer 最常见的三大用途，师兄给你背下来：

1. **关文件**：`f, _ := os.Open(...); defer f.Close()` ——打开立刻登记关闭，永远配对
2. **释放锁**：`mu.Lock(); defer mu.Unlock()` ——加锁立刻登记解锁，死锁概率骤降
3. **关闭网络连接/清理 goroutine 资源**：`conn, _ := net.Dial(...); defer conn.Close()`

新人最容易踩的坑是 **defer 参数的"立即求值"**。比如：

```go
x := 1
defer fmt.Println(x)   // 打印 1，不是 10
x = 10
```

defer 登记那一刻 x 就被当场求值为 1，后面再怎么改 x 都不影响。如果你想捕获"最新值"，得用闭包：`defer func(){ fmt.Println(x) }()`。

另外一个坑是 **循环里 defer**：for 循环里每轮都 defer，要等整个外层函数返回才集体执行——打开一万个文件的循环里写 defer，资源会撑爆。正确姿势是把一轮工作抽成独立函数，在那个函数里 defer。

🎓 知识点清单

- defer 是关键字，登记一个函数调用等外层函数 return 前执行
- defer 的参数在登记那一刻立即求值，不是真正执行时求值
- 多个 defer 按 LIFO（后进先出）顺序执行
- panic 触发栈展开时，沿途 defer 照常执行，所以 defer 是 panic-safe 的
- 典型用途：关文件、解锁、关连接，和资源打开成对写
- Close() 这类"结尾"操作也可能返回 error，defer 里依然要检查
- 循环里直接 defer 容易资源撑爆，抽成独立函数在函数里 defer

➡️ 下一关

defer 解决了"函数正常返回前清理"的问题，但如果 panic 真炸了，**怎么把程序从半空接住、让它继续跑下去**？这就是第 47 关 Recover 登场的时候了。recover 必须写在 defer 里才有用，三位一体的故事最后一环，咱们走起！
