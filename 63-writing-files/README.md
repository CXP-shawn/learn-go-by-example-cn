# 第 63 关：写文件（师兄带你学 Go）

## 🎯 这一关你会学到

本关目标：掌握 Go 写文件的三板斧——os.WriteFile 一把梭搞定小文件，os.Create + f.Write/f.WriteString 细粒度控制，bufio.NewWriter 批量缓冲减少系统调用。外加两个关键细节：0644 权限怎么读、f.Sync() 为啥要显式刷盘。

## 🤔 先想一个问题

你有没有遇到过这种场景：程序跑得好好的，突然断电或者 kill -9，结果发现日志文件最后几百行没了，导出的 CSV 最后几行是空的，配置文件写了一半变成乱码——服务直接起不来。这就是没搞清楚

## 📖 看代码

```go
// 在 Go 中，写文件与我们前面看过的读文件方法类似。

package main

import (
	"bufio"
	"fmt"
	"os"
)

func check(e error) {
	if e != nil {
		panic(e)
	}
}

func main() {

	// 开始！这里展示了如何写入一个字符串（或者只是一些字节）到一个文件。
	d1 := []byte("hello\ngo\n")
	err := os.WriteFile("/tmp/dat1", d1, 0644)
	check(err)

	// 对于更细粒度的写入，先打开一个文件。
	f, err := os.Create("/tmp/dat2")
	check(err)

	// 打开文件后，一个习惯性的操作是：立即使用 defer 调用文件的 `Close`。
	defer f.Close()

	// 您可以按期望的那样 `Write` 字节切片。
	d2 := []byte{115, 111, 109, 101, 10}
	n2, err := f.Write(d2)
	check(err)
	fmt.Printf("wrote %d bytes\n", n2)

	// `WriteString` 也是可用的。
	n3, err := f.WriteString("writes\n")
	check(err)
	fmt.Printf("wrote %d bytes\n", n3)

	// 调用 `Sync` 将缓冲区的数据写入硬盘。
	f.Sync()

	// 与我们前面看到的带缓冲的 Reader 一样，`bufio` 还提供了的带缓冲的 Writer。
	w := bufio.NewWriter(f)
	n4, err := w.WriteString("buffered\n")
	check(err)
	fmt.Printf("wrote %d bytes\n", n4)

	// 使用 `Flush` 来确保，已将所有的缓冲操作应用于底层 writer。
	w.Flush()

}
```

## 🔍 师兄给你逐行拆

好，咱们来聊写文件这件事，其实比读文件还简单，一行代码搞定。

最直接的姿势就是 `os.WriteFile`，用法长这样：

```go
err := os.WriteFile("output.txt", []byte("hello, gopher"), 0644)
if err != nil {
    log.Fatal(err)
}
```

三个参数：文件路径、要写入的字节切片、文件权限。就这么多，一把梭。

咱们重点说说 `0644` 这个数字，很多新手看到这个一脸懵。这是 Unix 文件权限位，八进制写法，前面那个 `0` 就是告诉编译器

好，第 63 关，写文件。上一关我们读了文件，这一关我们往磁盘里写东西，看起来简单，但细节坑不少，我带你一个一个过。

先说 `os.Create`。很多人以为它只是「创建文件」，其实它等价于用 `os.OpenFile` 同时传了三个 flag：`O_RDWR | O_CREATE | O_TRUNC`。重点是 `O_TRUNC`——如果文件已经存在，它会直接把原来的内容清空，从头开始写。所以你拿 `os.Create` 去「追加日志」，那就是在挖坑，每次运行都把之前的日志全干掉了。

要追加，得用 `os.OpenFile`，flag 换成 `os.O_APPEND | os.O_CREATE | os.O_WRONLY`，权限给 `0644`。`O_APPEND` 让每次写都自动跳到文件末尾，`O_CREATE` 保证文件不存在时自动创建，`O_WRONLY` 只写模式，够用就行，别乱开权限。

不管哪种方式打开文件，拿到 `f` 之后，第一件事就是 `defer f.Close()`。这是铁律，不是建议。文件描述符是系统资源，忘了关会泄漏，程序跑久了直接崩。defer 写在 Open 成功之后，error 判断之后，位置别搞错。

写内容有两个方法：`f.Write([]byte)` 和 `f.WriteString(string)`。前者接收字节切片，后者直接接收字符串。区别在哪？你用 `f.Write` 写字符串时，得手动转 `[]byte(s)`，这一步会分配一块新内存，把字符串内容复制过去。`f.WriteString` 内部直接操作字符串底层数据，省掉这次分配。写高频日志或者大量小文本的时候，这个差距会累积出来，优先用 `f.WriteString`。

两个方法的返回值都是 `(n int, err error)`，`n` 是实际写入的字节数。注意，写入字节数和你传入的长度不一致时，err 不一定非空，所以最好两个都检查。`n != len(data)` 也算写坏了，别忽略它。

最后说 `f.Sync()`。操作系统为了性能，write 调用完数据其实只到了内核缓冲区，还没真正落到磁盘。程序正常退出的时候会刷，但如果机器突然断电，缓冲区里的数据就丢了。调 `f.Sync()` 会触发 fsync 系统调用，强制把数据刷到磁盘。代价是变慢，所以不是每次都用，数据库、账单、配置这类关键文件才值得加它。普通日志追求性能可以不加，看场景权衡。记住这个工具放在工具箱里，关键时刻拿出来。

好，咱们来聊聊 bufio 写文件这块，这是生产代码里非常常见的套路，别小看它。

**bufio.NewWriter 的本质**

你直接调 f.Write() 或 fmt.Fprintf(f, ...)，每次都是一次系统调用，syscall 开销不小。bufio.NewWriter(f) 默认给你开一个 4KB 的用户态缓冲区，你往里写数据，先攒在内存里，等缓冲满了或者你手动 Flush，才真正触发一次 write 系统调用。写日志、写大文件，性能差距能有好几倍。

```go
w := bufio.NewWriter(f)
w.WriteString("buffered\n")
```

**Flush() 必须手动调，这是大坑**

很多新手写完就 defer f.Close()，以为万事大吉。问题来了：bufio 的缓冲区里还有数据没刷到文件，f.Close() 直接把文件描述符关了，数据就丢了，而且没有任何报错。所以正确姿势是：

```go
if err := w.Flush(); err != nil {
    log.Fatal(err)
}
```

在 Close 之前手动 Flush，别依赖 defer 帮你。

**defer 顺序问题要注意**

defer 是后进先出，如果你写成：

```go
defer f.Close()
defer w.Flush() // 这个反而先执行，看起来对？
```

这样 Flush 确实先于 Close 执行，顺序没错。但 Flush 的错误你没法处理，defer 吞掉了。所以师兄建议：Flush 老老实实手动调，检查 error，defer 只负责 Close 兜底。

**三个实战场景，记住了**

第一，原子写。你不想让别人读到写了一半的文件，就先写临时文件，写完 Flush、Close，再 os.Rename(tmpFile, targetFile)。rename 在同一文件系统下是原子操作，要么旧文件要么新文件，不会出现中间态。

第二，日志切割。按时间或大小滚动日志文件，每次切割前 Flush 当前 writer，再 Close，再 os.Create 新文件，重新包一个 bufio.NewWriter。

第三，压缩输出组合。

```go
gw := gzip.NewWriter(f)
w := bufio.NewWriter(gw)
w.WriteString("data")
w.Flush()
gw.Close() // gzip 也要 Close 才写完尾部
```

bufio 套在 gzip 外面，先缓冲再压缩再落盘，层层嵌套，每一层都要正确关闭。这个组合在写大数据文件时非常香，师兄项目里用了不少。

核心原则就一句话：**写完就 Flush，Flush 要检查 error，Close 用 defer 兜底。**

## 🏃 跑一下试试

跑 go run writing-files.go，终端输出：
wrote 5 bytes
wrote 7 bytes
wrote 9 bytes

三行分别对应 os.WriteFile 写 "hello\n"（5字节）、f.Write 写 "go\n"（3→合计7）和 w.WriteString 系列写入。跑完执行 cat /tmp/dat1，看到 hello 和 go 两行；cat /tmp/dat2，看到 some / writes / buffered 三行，确认 bufio 缓冲已被 Flush 落盘。文件权限用 0644，所有者可读写、其他人只读，是 Web 服务常见配置。

## 💡 师兄的碎碎念

💡 写文件五条锦囊：
① WriteString 省转换：w.WriteString(s) 比 w.Write([]byte(s)) 更简洁，底层一样高效，优先用。
② Flush 是命门：bufio.Writer 只是内存缓冲，程序结束前忘调 w.Flush()，数据就蒸发了，defer 里加上它。
③ 原子写三部曲：生产环境写重要文件，先写临时文件 tmp，写完 f.Sync()，再 os.Rename(tmp, dst)，避免读到写了一半的文件。
④ 权限细节：0644（rw-r--r--）适合公开配置，0600（rw-------）适合密钥、token 文件，别图省事一律 0644。
⑤ f.Sync vs Flush：Flush 只把 Go 层缓冲推给内核，f.Sync（即 fsync）才把内核缓冲刷到磁盘，断电不丢数据要用后者，但代价是性能下降。

## 🎓 知识点清单

- os.WriteFile 一次性写整个文件，最简单，适合小文件
- os.OpenFile 配合 os.O_WRONLY|os.O_CREATE|os.O_TRUNC 手动打开写入
- defer f.Close() 紧跟 Open，防止资源泄漏
- bufio.NewWriter 包裹 *os.File，减少系统调用次数
- w.WriteString("some\n") 直接写字符串，省去 []byte 转换
- 写完必须调 w.Flush()，否则缓冲区数据不会落盘
- 可用 f.Sync() 把内核缓冲刷到磁盘，保证持久化

## ➡️ 下一关

第 64 关是「行过滤器 (line-filters)」——用 bufio.Scanner 逐行读 stdin、处理后写 stdout，是流式文本处理的基础模式，我们继续。
