# 第 62 关：读文件（师兄带你学 Go）

## 🎯 这一关你会学到

掌握 Go 读文件的六种姿势：os.ReadFile 一次性读全、os.Open 获取文件句柄、f.Read 底层字节读取、f.Seek 定位读指针、io.ReadAtLeast 保底读取字节数，以及 bufio.NewReader + Peek 带缓冲读取，并了解各自适用场景与注意事项。

## 🤔 先想一个问题

你肯定遇过这种需求：程序启动时读一份 config.yaml 载入配置，或者分析一段 nginx access.log 找出慢请求，再或者解析一个 CSV 只看前几列。这些任务背后全是

## 📖 看代码

```go
// 读写文件在很多程序中都是必须的基本任务。
// 首先我们来看一些读文件的例子。

package main

import (
	"bufio"
	"fmt"
	"io"
	"os"
)

// 读取文件需要经常进行错误检查，
// 这个帮助方法可以精简下面的错误检查过程。
func check(e error) {
	if e != nil {
		panic(e)
	}
}

func main() {

	// 最基本的文件读取任务或许就是将文件内容读取到内存中。
	dat, err := os.ReadFile("/tmp/dat")
	check(err)
	fmt.Print(string(dat))

	// 您通常会希望对文件的读取方式和内容进行更多控制。
	// 对于这个任务，首先使用 `Open` 打开一个文件，以获取一个 `os.File` 值。
	f, err := os.Open("/tmp/dat")
	check(err)

	// 从文件的开始位置读取一些字节。
	// 最多允许读取 5 个字节，但还要注意实际读取了多少个。
	b1 := make([]byte, 5)
	n1, err := f.Read(b1)
	check(err)
	fmt.Printf("%d bytes: %s\n", n1, string(b1[:n1]))

	// 你也可以 `Seek` 到一个文件中已知的位置，并从这个位置开始读取。
	o2, err := f.Seek(6, 0)
	check(err)
	b2 := make([]byte, 2)
	n2, err := f.Read(b2)
	check(err)
	fmt.Printf("%d bytes @ %d: ", n2, o2)
	fmt.Printf("%v\n", string(b2[:n2]))

	// 例如，`io` 包提供了一个更健壮的实现 `ReadAtLeast`，用于读取上面那种文件。
	o3, err := f.Seek(6, 0)
	check(err)
	b3 := make([]byte, 2)
	n3, err := io.ReadAtLeast(f, b3, 2)
	check(err)
	fmt.Printf("%d bytes @ %d: %s\n", n3, o3, string(b3))

	// 没有内建的倒带，但是 `Seek(0, 0)` 实现了这一功能。
	_, err = f.Seek(0, 0)
	check(err)

	//  `bufio` 包实现了一个缓冲读取器，这可能有助于提高许多小读操作的效率，以及它提供了很多附加的读取函数。
	r4 := bufio.NewReader(f)
	b4, err := r4.Peek(5)
	check(err)
	fmt.Printf("5 bytes: %s\n", string(b4))

	// 任务结束后要关闭这个文件
	// （通常这个操作应该在 `Open` 操作后立即使用 `defer` 来完成）。
	f.Close()
}
```

## 🔍 师兄给你逐行拆

好，第62关——读文件。Go 处理文件读取的方式非常接近 POSIX 的底层风味，但又加了一层贴心的封装。我们一段一段拆开来聊。

**第一块：check 辅助函数**

```go
func check(e error) { if e != nil { panic(e) }
```

这个小函数就一件事：如果 error 不是 nil，直接 panic。教学示例里这样写是为了省掉每行 `if err != nil { ... }` 的重复噪音，让代码主干更清晰。但请注意——这是教学捷径，不是生产规范！生产代码里你应该认真处理每一个 error，把它向上返回或者包装后返回，而不是直接炸掉整个程序。panic 在服务里会让你的请求全挂掉，线上别这么干，师兄见过太多

好，继续拆第二段代码，上一节我们把文件打开、读出来了，这次来点更骚的操作——随机访问和缓冲读取。

**f.Seek：文件指针你说了算**

`f.Seek(6, 0)` 这一行，意思是「把读写指针移到距离文件头第 6 个字节的位置」。第二个参数 whence 有三个值：0 是从文件头算，1 是从当前位置算，2 是从文件末尾算。代码里用了两次 `Seek(6, 0)`，都是从头跳到第 6 字节，然后各自读 2 字节。为啥要读两次？因为作者想对比两种读法的区别，下面就来了。

第一种是直接 `f.Read(b2)`，这没啥好说的，老老实实读，但它不保证一定能读满 2 字节——如果文件剩余内容不够，它就给你读多少算多少，返回值 n2 告诉你实际读了几个，你自己去判断够不够，容易踩坑。

**io.ReadAtLeast：比 f.Read 更有担当**

第二种是 `io.ReadAtLeast(f, b3, 2)`，第三个参数 min=2，意思是「你必须给我至少读 2 字节，不然就报错」。如果文件内容不够 2 字节，它会返回 `io.ErrUnexpectedEOF`，让你一眼看出是数据不足，而不是默默少读。生产环境里解析二进制协议、网络包头，这个函数能帮你省掉不少防御性判断，强烈推荐。说白了，`f.Read` 是个佛系选手，`io.ReadAtLeast` 是个有底线的人。

**Seek(0, 0)：优雅倒带**

`f.Seek(0, 0)` 这一行注释写着「倒带」，非常形象。文件读完之后指针在末尾，你要重新从头读，就用这个姿势——偏移量 0，从文件头算，指针回到起点。这是 Go 里最通用的重置姿势，记住就行，别每次都去查文档。

**bufio.NewReader：加一层缓冲，少烦内核**

`bufio.NewReader(f)` 在文件外面包了一层缓冲读取器。为啥要这层？因为每次直接调 `f.Read` 都可能触发一次系统调用，频繁小块读取性能很差。bufio 会一次多读一大块到内存缓冲区，后续的小读直接从缓冲区取，系统调用次数大幅下降，性能蹭蹭涨。

**Peek：偷看不消耗**

`r4.Peek(5)` 是 bufio.Reader 独有的骚操作——往前看 5 字节，但是不移动指针，数据还在缓冲区里。这在判断文件类型、协议魔数的时候特别好用：先 Peek 看个头部，根据内容决定怎么解析，再正式 Read。如果用普通 Read，读完就没了，你还得想办法「退回去」，麻烦得很。

**延伸一秒钟**

逐行读文件用 `bufio.Scanner`，它会自动处理换行符，比你手动切 `\n` 舒服很多。大文件从一个地方搬到另一个地方，用 `io.Copy`，流式拷贝，不会把整个文件塞进内存，内存友好型选手。这两个后面都会遇到，先记个名字。

读文件这件事，Go 处理得其实挺顺手，但坑也不少，来聊聊实战里最常见的几种姿势。

**小文件直接一把梭：os.ReadFile**

读个配置文件、JSON？别搞那么多仪式感，一行搞定：`data, err := os.ReadFile("config.json")`。返回 []byte，配合 json.Unmarshal 直接解析，干净利落。但记住，这是把整个文件塞进内存的，几十 MB 的日志你敢这么读，OOM 了别找我哭。

**大文件逐行读：bufio.Scanner**

日志分析、CSV 处理，必须上 bufio.Scanner。标准写法是：
```
f, _ := os.Open("big.log")
defer f.Close()
scanner := bufio.NewScanner(f)
for scanner.Scan() {
    line := scanner.Text()
    // 处理每一行
}
```
这里有个经典翻车现场——默认 buffer 只有 64KB，遇到超长行直接报 `token too long`。解决方案是手动扩容：`scanner.Buffer(make([]byte, 1024*1024), 1024*1024)`，调大 buffer 就行了，别死守默认值。

**二进制流式处理：io.Copy**

下载文件写磁盘、边读边算 MD5？`io.Copy(dst, src)` 是你的好朋友。它内部自带 buffer，不会把整个文件吃进内存，接 hash.Writer 或者 net.Conn 都毫无压力，优雅得很。

**编码坑，Windows 用户专属惊喜**

从 Windows 来的文件有两个常见地雷：一是 UTF-8 BOM（文件头多了 `\xEF\xBB\xBF` 三个字节），解析 JSON 直接报错，得手动 trim 掉；二是 CRLF 换行（`\r\n`），bufio.Scanner 默认按 `\n` 切，行尾会残留 `\r`，字符串比较全错。遇到这种情况，`strings.TrimRight(line, "\r")` 保平安。

**defer f.Close()，这条是铁律**

Open 之后立刻写 defer f.Close()，手别抖，不写就等着文件描述符泄漏吧。高并发场景下，fd 耗尽的报错能让你怀疑人生。养成习惯，open 和 defer 就是亲兄弟，形影不离。

## 🏃 跑一下试试

先造测试文件，再跑程序：

```
echo "hello go" > /tmp/dat
go run reading-files.go
```

预期输出共 5 行：

```
hello go          ← os.ReadFile 读全文
5 bytes: hello    ← Read(b) 读前 5 字节
2 bytes @ 6: go   ← Seek 到偏移 6 再读 2 字节
2 bytes @ 6: go   ← bufio.NewReader + Seek 重复验证
5 bytes: hello    ← bufio.Reader.Read 读前 5 字节
```

`echo` 会自动追加换行符，所以文件内容是 `hello go\n`，共 9 字节。偏移 6 正好指向 `g`，读 2 字节得到 `go`。若输出对不上，先用 `xxd /tmp/dat` 确认字节布局。

## 💡 师兄的碎碎念

💡 **读文件避坑指南**

1. **defer f.Close() 铁律**：`os.Open` 成功后下一行就写 `defer f.Close()`，别等函数末尾手动关，忘了就是文件描述符泄漏。

2. **bufio.Scanner 长行陷阱**：Scanner 默认 buffer 上限 64 KB，日志行超长时会返回 `bufio.ErrTooLong`。超长场景改用 `scanner.Buffer(make([]byte, 1<<20), 1<<20)` 扩容，或直接换 `bufio.Reader.ReadLine`。

3. **io.ReadAll 替代 ioutil.ReadAll**：Go 1.16 起 `ioutil` 已废弃，读全部内容用 `io.ReadAll(r)`，小文件也可一步到位用 `os.ReadFile(path)` 省去 Open/Close。

4. **Read(b) 不保证填满 b**：底层 `Read` 返回实际读到的字节数 `n`，只用 `b[:n]` 才安全；需要填满就用 `io.ReadFull`。

5. **Windows BOM & CRLF**：Windows 文本文件可能带 UTF-8 BOM（`EF BB BF`）和 `\r\n` 换行，跨平台脚本记得用 `bytes.TrimPrefix` 去 BOM、`strings.ReplaceAll` 处理 CRLF，避免解析结果莫名多出空字符。

## 🎓 知识点清单

- echo "hello go" > /tmp/dat 准备好测试文件
- go run reading-files.go 输出 5 行与预期一致
- defer f.Close() 紧跟 os.Open() 防止资源泄漏
- Read(b) 返回值 n 已正确使用，未忽略
- Seek 偏移量计算正确（字节偏移，非字符偏移）
- bufio.NewReader 包裹 *os.File 以享受缓冲读取
- io.ReadAll / os.ReadFile 在适当场景替换低级 API
- 代码在 Windows 上注意 CRLF 与 BOM 差异

## ➡️ 下一关

📂 读文件掌握了，翻硬币到另一面——**写文件 (writing-files)**！下一关你将用 os.Create、bufio.Writer、fmt.Fprintln 把数据落盘，还会看到 defer + Flush() 这对好搭档。准备好往磁盘里写字了吗？
