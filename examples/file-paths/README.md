# 第 65 关：文件路径（师兄带你学 Go）

## 🎯 这一关你会学到

掌握 path/filepath 包：用 Join 跨平台拼路径，Dir/Base/Split 拆分目录与文件名，IsAbs 判绝对路径，Ext 取扩展名，Rel 算相对路径，同时理解它相对 path 包的差异与规范化魔力。

## 🤔 先想一个问题

你写的工具在 Mac 上跑得好好的，丢到 Windows 同事电脑上立刻翻车——路径分隔符不对、路径里多了几个多余斜杠、绝对路径被当成相对路径解析……跨平台路径问题是新人最常见的坑，标准库的 path/filepath 就是来救命的。

## 📖 看代码

```go
// `filepath` 包为 *文件路径* ，提供了方便的跨操作系统的解析和构建函数；
// 比如：Linux 下的 `dir/file` 和 Windows 下的 `dir\file` 。

package main

import (
	"fmt"
	"path/filepath"
	"strings"
)

func main() {

	// 应使用 `Join` 来构建可移植(跨操作系统)的路径。
	// 它接收任意数量的参数，并参照传入顺序构造一个对应层次结构的路径。
	p := filepath.Join("dir1", "dir2", "filename")
	fmt.Println("p:", p)

	// 您应该总是使用 `Join` 代替手动拼接 `/` 和 `\`。
	// 除了可移植性，`Join` 还会删除多余的分隔符和目录，使得路径更加规范。
	fmt.Println(filepath.Join("dir1//", "filename"))
	fmt.Println(filepath.Join("dir1/../dir1", "filename"))

	// `Dir` 和 `Base` 可以被用于分割路径中的目录和文件。
	// 此外，`Split` 可以一次调用返回上面两个函数的结果。
	fmt.Println("Dir(p):", filepath.Dir(p))
	fmt.Println("Base(p):", filepath.Base(p))

	// 判断路径是否为绝对路径。
	fmt.Println(filepath.IsAbs("dir/file"))
	fmt.Println(filepath.IsAbs("/dir/file"))

	filename := "config.json"

	// 某些文件名包含了扩展名（文件类型）。
	// 我们可以用 `Ext` 将扩展名分割出来。
	ext := filepath.Ext(filename)
	fmt.Println(ext)

	// 想获取文件名清除扩展名后的值，请使用 `strings.TrmSuffix`。
	fmt.Println(strings.TrimSuffix(filename, ext))

	// `Rel` 寻找 `basepath` 与 `targpath` 之间的相对路径。
	// 如果相对路径不存在，则返回错误。
	rel, err := filepath.Rel("a/b", "a/b/t/file")
	if err != nil {
		panic(err)
	}
	fmt.Println(rel)

	rel, err = filepath.Rel("a/b", "a/c/t/file")
	if err != nil {
		panic(err)
	}
	fmt.Println(rel)
}
```

## 🔍 师兄给你逐行拆

好，今天我们来聊聊 Go 标准库里一个非常实用但经常被新人忽视的包——path/filepath。别看它名字朴素，真正做跨平台开发的时候，你会发现它简直是救命稻草。

先说说跨平台的痛点。咱们写代码的人，有人用 macOS，有人用 Linux，有人用 Windows，团队里三种系统都有很正常。问题来了：Linux 和 macOS 用正斜杠作为路径分隔符，比如 /home/user/documents/file.txt，而 Windows 用反斜杠，比如 C:\Users\user\documents\file.txt。如果你图省事，直接在代码里手工拼字符串，比如写成 dir + "/" + filename，这段代码在 Linux 上跑得好好的，一到 Windows 上就翻车了。更惨的是，有些同学反过来，写成 dir + "\\" + filename，结果 Linux 上又出问题。这种硬编码分隔符的做法，是跨平台开发的大忌，迟早给自己挖坑。

这时候 filepath.Join 就登场了。它是整个包里你最常用的函数，没有之一。用法非常简单，你把需要拼在一起的路径片段一个个传进去，它会自动用当前操作系统的原生分隔符把它们连起来，返回一个完整路径字符串。不管你在 Linux 还是 Windows 上跑，分隔符的选择都交给 Go 运行时去判断，你完全不用操心。代码写一份，到处跑，这才叫优雅。

更厉害的地方在于，filepath.Join 还会自动帮你规范化路径。什么叫规范化？就是它会把路径里那些乱七八糟的地方都收拾干净。比如你不小心传进去的路径片段里带了多余的斜杠，或者带了表示当前目录的点号，甚至带了表示上级目录的两个点，它都会帮你正确处理。举个具体例子：如果你把 dir1、// 和 filename 三个字符串拼在一起，它不会傻乎乎地给你返回 dir1//filename 这种奇怪的结果，而是规规矩矩地返回 dir1/filename，多余的斜杠被压缩掉了。再比如你传入 /home/user/./documents 这样的路径，里面的点号会被解析掉，得到干净的 /home/user/documents。碰到两个点的上级目录引用也一样，/home/user/../documents 会被解析成 /home/documents。这个自动规范化的能力，省去了你大量手动处理路径字符串的麻烦，少写很多容易出 bug 的字符串操作代码。

除了拼路径，filepath 包还提供了几个用来拆路径的函数，同样很常用。filepath.Dir 用来取一个路径的父目录部分，也就是去掉最后一段文件名或目录名之后剩下的部分。比如传入 /home/user/documents/file.txt，它返回 /home/user/documents。filepath.Base 正好相反，它取的是路径里最后一段，也就是文件名那部分，同样以 /home/user/documents/file.txt 为例，它返回 file.txt。如果你懒得分两次调用，还有一个 filepath.Split，它一次性同时返回目录部分和文件名部分，两个返回值，非常方便。平时你需要把一个完整路径拆开来分别处理的时候，这几个函数用起来特别顺手。

最后要特别提醒一件容易踩坑的事：Go 标准库里有两个长得很像的包，一个叫 path，另一个叫 path/filepath，它们是两个完全不同的东西，千万别混用。path 包只认识正斜杠，它处理的是纯逻辑意义上的路径，不感知操作系统。你处理 URL、HTTP 路由、某些纯逻辑层的路径拼接时，用 path 包是合适的，因为 URL 规范本来就用正斜杠，跟操作系统无关。但只要你在处理真实的文件系统路径，涉及到读写文件、遍历目录这类操作，就必须用 path/filepath，因为只有它才真正感知当前操作系统，才能给你正确的本地路径格式。

总结一句话：写跨平台的文件路径处理代码，忘掉手工拼字符串，忘掉 path 包，老老实实用 path/filepath，你的代码会干净很多，跨平台问题也会少很多。

好，段 B 继续！上一段把 Join、Split、Dir、Base 都收拾干净了，这一段我们来搞几个同样高频但容易被忽视的工具函数。还是那句话，标准库给你准备好了，别自己写字符串拼接——那条路通向 bug 的深渊。

**1. filepath.IsAbs：判断是不是绝对路径**

这个函数很简单，返回 bool。规则就一条：有没有前导分隔符（或者 Windows 上的盘符前缀）。

```go
fmt.Println(filepath.IsAbs("/etc/hosts"))   // true
fmt.Println(filepath.IsAbs("etc/hosts"))    // false
fmt.Println(filepath.IsAbs("./etc/hosts"))  // false
```

注意，`./etc/hosts` 也是相对路径，有点号也不算绝对。这在处理用户输入的路径时非常有用——先判断，再决定要不要拼 base 目录。

**2. filepath.Ext：提取扩展名（含点）**

```go
ext := filepath.Ext("config.json")  // ".json"
ext2 := filepath.Ext("archive.tar.gz")  // ".gz"
ext3 := filepath.Ext("README")  // ""
```

注意两点：第一，返回的扩展名**包含那个点**，别忘了；第二，对于 `archive.tar.gz` 这种双扩展名，只取最后一段 `.gz`，想拿完整的 `.tar.gz` 得自己再处理一次。这是吐槽点之一，新手经常在这里踩坑，以为能一次拿到全部后缀。

**3. 去掉扩展名：TrimSuffix**

`filepath` 包没有专门的

## 🏃 跑一下试试

逐行看输出：① p: dir1/dir2/filename —— Join 把三段拼成规范路径，自动去掉多余斜杠；② dir1/filename —— Join 只传两段；③ dir1/filename —— 同上，结果一致；④ Dir(p): dir1/dir2 —— Dir 返回最后一个元素之前的目录部分；⑤ Base(p): filename —— Base 返回路径最后一个元素即文件名；⑥ false —— dir1/dir2/filename 是相对路径，IsAbs 返回 false；⑦ true —— /home/user/file 以 / 开头，IsAbs 返回 true；⑧ .json —— Ext 提取 config.json 的扩展名，含点；⑨ config —— 把扩展名去掉后剩余的文件名主体；⑩ t/file —— Rel("a/b", "a/b/t/file") 计算相对路径；⑪ ../c/t/file —— Rel("a/b", "a/c/t/file") 需要先回退一层再进入 c。

## 💡 师兄的碎碎念

💡 Join 的规范化魔力：filepath.Join 不只是字符串拼接，它内部调用 Clean，会合并连续斜杠、处理 . 和 ..，让路径始终整洁，避免手动拼接带来的 bug。

💡 path vs filepath：标准库有两个路径包。path 只用正斜杠，适合 URL 或纯逻辑路径；filepath 感知操作系统，Windows 下自动用反斜杠。跨平台代码务必用 filepath，别混用。

💡 Windows 反斜杠：在 Windows 上 filepath.Join 返回 dir1\dir2\filename，直接硬编码斜杠会导致路径解析失败，用 Join 或 filepath.FromSlash 做转换。

💡 filepath.Clean 显式规范化：收到外部传入的路径（用户输入、配置文件）时，主动调用 Clean 防止路径穿越攻击（如 ../../etc/passwd）。

💡 Rel 跨盘符报错：Windows 上 filepath.Rel("C:\\a", "D:\\b") 会返回错误，因为两个盘符没有公共根，使用前需要检查 error。

## 🎓 知识点清单

- 理解 filepath.Join 自动规范化路径分隔符并清除多余斜杠
- 区分 path 包（URL、Unix 风格）与 filepath 包（操作系统原生路径）
- 掌握 filepath.Dir 取目录部分、filepath.Base 取文件名部分
- 用 filepath.IsAbs 判断绝对路径与相对路径
- 用 filepath.Ext 提取扩展名（含点），无扩展名时返回空字符串
- 用 filepath.Rel 计算两个路径的相对关系，注意跨盘符会返回错误
- 了解 filepath.Clean 显式规范化：消除 . / .. / 双斜杠等冗余元素

## ➡️ 下一关

路径搞定了，下一关「目录 (Directories)」我们直接上手：用 os.Mkdir、os.MkdirAll 创建目录，用 os.ReadDir 列出目录内容，用 filepath.WalkDir 递归遍历整棵目录树——文件系统操作从
