# 第 45 关：Panic（师兄带你学 Go）

🎯 这一关你会学到
• 什么是 panic——Go 的内置函数，表示"我没法继续了"
• panic 触发后会发生啥：打印 panic value、goroutine 栈、以退出码 2 结束
• panic 和日常 error 错误处理的边界在哪儿
• panic 会触发"栈展开"，沿途的 defer 还能执行（下一关就讲 defer）

🤔 先想一个问题

你用外卖 APP 的时候有没有遇到过这种情况——点完餐、付完钱、满心期待，突然屏幕啪一下黑了，APP 闪退回桌面，连个提示都没有？或者家里燃气报警器突然滴滴狂叫，邻居大爷扯着嗓子让全楼疏散？这俩场景有个共同点：事态严重到系统判断"必须立刻停下，不能再往下走了"。Go 里的 panic 就是这个角色——咱们师兄管它叫"紧急停止按钮"。按下去之后，当前这条执行路径就地报废，谁也拦不住。这一关咱们就把 panic 到底干了啥搞清楚。

## 📖 看代码

```go
// `panic` 意味着有些出乎意料的错误发生。
// 通常我们用它来表示程序正常运行中不应该出现的错误，
// 或者我们不准备优雅处理的错误。

package main

import "os"

func main() {

	// 我们将使用 panic 来检查这个站点上预期之外的错误。
	// 而该站点上只有一个程序：触发 panic。
	panic("a problem")

	// panic 的一种常见用法是：当函数返回我们不知道如何处理（或不想处理）的错误值时，中止操作。
	// 如果创建新文件时遇到意外错误该如何处理？这里有一个很好的 `panic` 示例。
	_, err := os.Create("/tmp/file")
	if err != nil {
		panic(err)
	}
}
```

🔍 师兄给你逐行拆

**panic 是个"内置函数"，跟 len、make 是一家子**

咱们先把 panic 的身份搞清楚。你翻遍 Go 的标准库目录，找不到 panic 这个函数住在哪个包里——因为它根本不在任何包里。panic 是 Go 的**内置函数（built-in function）**。所谓内置函数，就是编译器直接塞进语言里、不需要 import 任何东西就能用的函数。你打开一个空白的 .go 文件，啥都没 import，直接写 panic("炸了")，编译器认它。和它同门的还有 len、cap、make、new、append、copy、close、delete 这些老熟人，全都是不用 import 就能用。师兄我一般把它们类比成游戏里默认解锁的初始技能，不用点技能点，注册账号就能放。

panic 接收一个参数，类型是 any（Go 1.18 之前写作 interface{}），也就是说你传啥都行——字符串、整数、error 对象、自定义结构体，runtime 通通能收。它也没有返回值，因为调完之后整个执行流就变了样，不会走正常的 return 路径。

**panic("a problem") 这行跑下去发生了仨事**

看代码：`panic("a problem")`。这一行一旦被执行到，runtime（Go 运行时系统，负责调度、内存管理、GC 那些底层工作）立刻干三件事：

第一件，把你传进去的这个值——"a problem"——格式化之后打到 stderr（标准错误输出流）上。你在终端会看到开头一行 `panic: a problem`。传字符串就直接打字符串；传 error 类型，runtime 会帮你调 .Error() 方法拿字符串；传别的任意类型，runtime 会用类似 %v 的方式尽量格式化出来。

第二件，把当前 goroutine 的**调用栈（call stack）**打印出来。goroutine 是 Go 的轻量级线程，你可以理解成一种超便宜、开销远小于系统线程的并发执行单位，main 函数跑在编号 1 的 goroutine 里。所谓调用栈，就是"我现在这一行代码，是从哪儿一步步被调过来的"——像快递单上的物流轨迹：顺丰深圳分拣中心→广州中转→用户手里，一层层写清楚。你能看到类似 `goroutine 1 [running]`、`main.main()`、`panic.go:某行` 的信息，对着文件名+行号一眼就能定位到事发现场。

第三件，**以退出码 2 结束进程**。Go runtime 对"没被 recover 接住的 panic"固定返回退出码 2，正常结束是 0。CI 脚本、shell 命令链（bash 里 `$?` 就是上一条命令的退出码）都靠这个码判断你到底是正常退出还是炸了。

**栈展开（unwinding）：panic 不是瞬间消失，是层层上爬**

这里要引入一个重要概念——**栈展开（unwinding）**。panic 不是啪一下就让进程消失的，而是从触发的那一层开始，沿着调用栈一层层往上返回：先退出当前函数、再退出调用者、再退出调用者的调用者……直到退到 main 函数顶层没人再接了，整个进程才挂掉。每退一层，runtime 都会检查那一层有没有用 defer 注册过延迟函数——如果有，就挨个执行一遍。咱们这一关源码里还没用到 defer，所以 panic 直接一路爬到顶，程序就噶了。下一关你就能看到 defer 怎么跟 panic 配合，再下一关 recover 还能把 panic 从半空接住。

**第二段 os.Create 那坨代码为啥永远执行不到？**

看源码下半段：
```go
_, err := os.Create("/tmp/file")
if err != nil {
    panic(err)
}
```

这段代码演示的是另一种典型姿势：调了某个可能返回 error 的函数（os.Create 创建文件），如果 err 不是 nil（nil 在 Go 里表示"空"，约等于别的语言的 null），就用 panic 把 err 一把扔出去。但在咱们这个源码里，这段永远跑不到——因为上面那行 panic("a problem") 已经把当前 goroutine 的执行流打断了，栈展开过程中根本不会回到 main 继续往下走。师兄我一开始看这段也困惑，后来才明白：原站把这段放这儿纯粹是**演示用法**，告诉你"如果你想这么写，可以这么写"，并不是真想让它执行。

**panic 是最后的求救信号，不是日常错误处理**

重点来了。Go 的错误处理哲学跟 Java/Python 的 exception 体系完全不是一路。**error 是内置接口**（定义：任何实现了 `Error() string` 方法的类型都算 error），它是一等公民，函数返回值里经常带 error；调用方用 `if err != nil` 分支决定怎么处理——记日志、降级、重试、原样返回给上层。这才是 Go 里的常态。

panic 留给啥场景？留给真正**不可恢复**的情况：启动时加载配置文件失败、某个全局不变量被破坏（比如红黑树居然不平衡了）、明显的编程 bug（如数组越界 runtime 会自己 panic）。新手最容易犯的错是把 panic 当 Java 的 throw 用，到处扔——库代码里这么干极度不礼貌，调用方根本不知道你会不会炸，还得满屏加 defer+recover 防你，烦死人。师兄我的原则是：**应用入口可以 panic，库代码老老实实 return err**。

最后剧透一下：panic 并不是完全无法阻挡。下一关咱们讲 defer，defer 注册的函数在栈展开时依然会执行；再下一关讲 recover，它能在 defer 里把飞过来的 panic 接住，让程序苟活下来。这俩是 panic 的亲兄弟，三位一体。

🏃 跑一下试试

保存为 panic.go，然后：

```bash
go run panic.go
```

你会看到类似这样的输出：

```text
panic: a problem

goroutine 1 [running]:
main.main()
	/你的路径/panic.go:10 +0x27
exit status 2
```

逐段拆解：
- `panic: a problem` —— 你传给 panic() 的值，string 直接打出来
- `goroutine 1 [running]` —— 在 main goroutine 里（编号固定 1）出的事，状态 running 说明就是跑着跑着炸的
- `main.main() /.../panic.go:10 +0x27` —— 调用栈，文件名+行号定位到事发点，+0x27 是函数内的指令偏移量，平时不用管
- `exit status 2` —— 未被 recover 的 panic 固定退出码 2，CI 脚本可以靠它识别崩溃

💡 师兄的碎碎念

新人最容易犯的误区：**把 panic 当 Java/Python 的 throw 用，到处扔**。师兄我求求你别这么干。

Go 的错误处理哲学是"能 error 就 error"。你写一个库函数，返回 (T, error) 是礼貌的做法——调用方自己决定怎么处理。你要是在库里随手 panic，调用方得满屏 defer+recover 防你，烦不烦？标准库里 json.Marshal 遇错也是 return err，不是 panic，这就是教科书级的姿势。

那 panic 啥时候能用？**应用入口初始化失败可以 panic**：比如启动时读配置文件失败、连数据库连不上、必要的环境变量没配——这些情况下程序本来就没法正常工作，panic 直接炸给 ops 看，总比带着残缺状态继续跑要安全。还有就是**明显违反不变量的情况**：比如你写了个排序算法，排完发现没排好，这是 bug，panic 比装没事发生强。

顺便一提，runtime 自己也会主动 panic——数组越界、nil 指针解引用、map 并发写、除以零，这些都会触发 runtime panic。所以你代码里看不到 panic 关键字也可能炸，别觉得没写就安全。

🎓 知识点清单

- panic 是 Go 的内置函数，不用 import 就能直接调用
- panic 接收一个 any 参数，可以传任意值
- panic 触发后：向 stderr 打印 panic value、打印当前 goroutine 栈、以退出码 2 结束进程
- panic 会触发栈展开（unwinding），沿途执行 defer
- 源码里 panic 后面的代码永远跑不到
- error 才是 Go 日常错误处理的主角，panic 留给真正不可恢复的极端情况
- runtime 自身也会在数组越界、nil 解引用、map 并发写等情况下主动 panic
- 下一关 defer + 再下一关 recover 才是 panic 的兜底方案

➡️ 下一关

既然 panic 会触发栈展开，沿途的 defer 还有机会执行——那 defer 到底是怎么登记的？执行顺序是啥？参数是什么时候求值的？下一关第 46 关「Defer」咱们一次拆清楚，走起！
